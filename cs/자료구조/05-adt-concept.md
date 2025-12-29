# 추상 데이터 타입 (Abstract Data Type, ADT)

## 개요

추상 데이터 타입(ADT)은 데이터와 그 데이터에 대한 연산을 **구현 세부사항과 분리**하여 정의한 수학적 모델이다. ADT는 "무엇을(What)" 하는지만 명시하고, "어떻게(How)" 하는지는 숨긴다. 이를 통해 인터페이스와 구현을 분리하여 유연하고 유지보수 가능한 코드를 작성할 수 있다.

## 핵심 개념

### ADT의 구성 요소

```
ADT = (도메인, 연산, 공리)

1. 도메인 (Domain): 데이터의 집합
2. 연산 (Operations): 데이터에 대한 동작
3. 공리 (Axioms): 연산의 의미를 정의하는 규칙
```

### Interface vs Implementation

```
┌─────────────────────────────────────────────────────────┐
│                      Client Code                        │
│         (ADT의 연산만 사용, 구현 무관)                    │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    Interface (ADT)                      │
│   - 연산의 시그니처 (입력/출력 타입)                       │
│   - 연산의 의미 (사전/사후 조건)                          │
│   - 시간/공간 복잡도 보장                                │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    Implementation                       │
│   - 구체적인 자료구조 선택                                │
│   - 알고리즘 구현                                        │
│   - 메모리 관리                                          │
└─────────────────────────────────────────────────────────┘
```

### Specification vs Implementation

**Specification (명세)**: ADT가 "무엇을" 하는지 정의

```java
/**
 * Stack ADT 명세
 *
 * 도메인: 원소들의 LIFO(Last-In-First-Out) 컬렉션
 *
 * 연산:
 *   - push(e): 원소 e를 스택 맨 위에 추가
 *   - pop(): 스택 맨 위 원소를 제거하고 반환
 *   - top()/peek(): 스택 맨 위 원소를 반환 (제거하지 않음)
 *   - isEmpty(): 스택이 비어있으면 true
 *   - size(): 원소의 개수 반환
 *
 * 공리:
 *   - pop(push(S, e)) = S (push 후 pop하면 원래 스택)
 *   - top(push(S, e)) = e (push한 원소가 맨 위)
 *   - isEmpty(new Stack()) = true
 *   - isEmpty(push(S, e)) = false
 */
public interface Stack<E> {
    void push(E element);
    E pop();
    E peek();
    boolean isEmpty();
    int size();
}
```

**Implementation (구현)**: ADT를 "어떻게" 구현하는지 정의

```java
// 구현 1: 배열 기반
public class ArrayStack<E> implements Stack<E> {
    private E[] array;
    private int top;
    private static final int DEFAULT_CAPACITY = 10;

    @SuppressWarnings("unchecked")
    public ArrayStack() {
        array = (E[]) new Object[DEFAULT_CAPACITY];
        top = -1;
    }

    @Override
    public void push(E element) {
        if (top == array.length - 1) {
            resize(2 * array.length);
        }
        array[++top] = element;
    }

    @Override
    public E pop() {
        if (isEmpty()) throw new EmptyStackException();
        E element = array[top];
        array[top--] = null;  // 메모리 누수 방지
        return element;
    }

    @Override
    public E peek() {
        if (isEmpty()) throw new EmptyStackException();
        return array[top];
    }

    @Override
    public boolean isEmpty() {
        return top == -1;
    }

    @Override
    public int size() {
        return top + 1;
    }

    private void resize(int newCapacity) {
        array = Arrays.copyOf(array, newCapacity);
    }
}

// 구현 2: 연결 리스트 기반
public class LinkedStack<E> implements Stack<E> {
    private Node<E> head;
    private int size;

    private static class Node<E> {
        E data;
        Node<E> next;

        Node(E data, Node<E> next) {
            this.data = data;
            this.next = next;
        }
    }

    @Override
    public void push(E element) {
        head = new Node<>(element, head);
        size++;
    }

    @Override
    public E pop() {
        if (isEmpty()) throw new EmptyStackException();
        E element = head.data;
        head = head.next;
        size--;
        return element;
    }

    @Override
    public E peek() {
        if (isEmpty()) throw new EmptyStackException();
        return head.data;
    }

    @Override
    public boolean isEmpty() {
        return head == null;
    }

    @Override
    public int size() {
        return size;
    }
}
```

### 주요 ADT와 구현

#### 1. List ADT

```java
/**
 * List ADT 명세
 *
 * 도메인: 순서가 있는 원소들의 컬렉션
 *
 * 연산:
 *   - get(i): i번째 원소 반환
 *   - set(i, e): i번째 원소를 e로 변경
 *   - add(i, e): i번째 위치에 e 삽입
 *   - remove(i): i번째 원소 제거
 *   - size(): 원소 개수
 */
public interface List<E> {
    E get(int index);
    void set(int index, E element);
    void add(int index, E element);
    E remove(int index);
    int size();
}
```

| 구현 | get | set | add(중간) | remove(중간) | 메모리 |
|------|-----|-----|-----------|--------------|--------|
| ArrayList | O(1) | O(1) | O(n) | O(n) | 연속적 |
| LinkedList | O(n) | O(n) | O(1)* | O(1)* | 분산 |

*위치를 찾은 후의 연산, 찾는 데 O(n) 필요

#### 2. Map ADT (Dictionary, Associative Array)

```java
/**
 * Map ADT 명세
 *
 * 도메인: (키, 값) 쌍의 컬렉션, 키는 유일
 *
 * 연산:
 *   - put(k, v): 키 k에 값 v를 매핑
 *   - get(k): 키 k에 매핑된 값 반환
 *   - remove(k): 키 k와 그 매핑 제거
 *   - containsKey(k): 키 k가 존재하면 true
 *   - size(): 매핑의 개수
 */
public interface Map<K, V> {
    V put(K key, V value);
    V get(K key);
    V remove(K key);
    boolean containsKey(K key);
    int size();
}
```

| 구현 | put | get | remove | 순서 보장 |
|------|-----|-----|--------|-----------|
| HashMap | O(1) avg | O(1) avg | O(1) avg | 없음 |
| TreeMap | O(log n) | O(log n) | O(log n) | 키 순서 |
| LinkedHashMap | O(1) avg | O(1) avg | O(1) avg | 삽입 순서 |

#### 3. Set ADT

```java
/**
 * Set ADT 명세
 *
 * 도메인: 중복 없는 원소들의 컬렉션
 *
 * 연산:
 *   - add(e): 원소 e 추가 (이미 있으면 무시)
 *   - remove(e): 원소 e 제거
 *   - contains(e): 원소 e가 있으면 true
 *   - size(): 원소 개수
 *   - union(S): 합집합
 *   - intersection(S): 교집합
 *   - difference(S): 차집합
 */
public interface Set<E> {
    boolean add(E element);
    boolean remove(E element);
    boolean contains(E element);
    int size();
}
```

| 구현 | add | remove | contains | 순서 |
|------|-----|--------|----------|------|
| HashSet | O(1) avg | O(1) avg | O(1) avg | 없음 |
| TreeSet | O(log n) | O(log n) | O(log n) | 정렬 |
| LinkedHashSet | O(1) avg | O(1) avg | O(1) avg | 삽입 순서 |

#### 4. Priority Queue ADT

```java
/**
 * Priority Queue ADT 명세
 *
 * 도메인: 우선순위가 있는 원소들의 컬렉션
 *
 * 연산:
 *   - insert(e, p): 우선순위 p로 원소 e 삽입
 *   - extractMin/Max(): 최소/최대 우선순위 원소 제거 후 반환
 *   - peekMin/Max(): 최소/최대 우선순위 원소 반환 (제거 안함)
 *   - changePriority(e, p): 원소 e의 우선순위를 p로 변경
 *   - isEmpty(): 비어있으면 true
 */
public interface PriorityQueue<E> {
    void insert(E element);
    E extractMin();
    E peekMin();
    boolean isEmpty();
}
```

| 구현 | insert | extractMin | peekMin | changePriority |
|------|--------|------------|---------|----------------|
| Binary Heap | O(log n) | O(log n) | O(1) | O(log n) |
| Fibonacci Heap | O(1) amortized | O(log n) amortized | O(1) | O(1) amortized |
| 정렬되지 않은 배열 | O(1) | O(n) | O(n) | O(1) |

### ADT의 장점

#### 1. 정보 은닉 (Information Hiding)

```java
// 클라이언트는 구현 세부사항을 몰라도 됨
public void processData(List<String> data) {
    // ArrayList든 LinkedList든 상관없이 동작
    for (int i = 0; i < data.size(); i++) {
        String item = data.get(i);
        // ...
    }
}
```

#### 2. 구현 교체 용이

```java
// 요구사항 변경 시 구현만 교체
// 변경 전: 빈번한 조회가 필요해서 ArrayList 사용
List<String> items = new ArrayList<>();

// 변경 후: 빈번한 삽입/삭제로 LinkedList로 교체
List<String> items = new LinkedList<>();

// 클라이언트 코드는 변경 불필요
```

#### 3. 테스트 용이

```java
// 인터페이스를 통한 목(Mock) 객체 생성
public class MockStack<E> implements Stack<E> {
    private List<String> operations = new ArrayList<>();

    @Override
    public void push(E element) {
        operations.add("push: " + element);
    }

    // 테스트 검증용
    public List<String> getOperations() {
        return operations;
    }
}
```

### 자바 컬렉션 프레임워크의 ADT

```
                    Collection
                        │
          ┌─────────────┼─────────────┐
          │             │             │
         List          Set          Queue
          │             │             │
    ┌─────┴─────┐  ┌────┴────┐   ┌────┴────┐
    │           │  │         │   │         │
ArrayList  LinkedList  HashSet  TreeSet  PriorityQueue


                       Map
                        │
          ┌─────────────┼─────────────┐
          │             │             │
       HashMap      TreeMap     LinkedHashMap
```

## 실무 적용

### ADT 선택 가이드

```java
// 1. 순서가 중요하고 인덱스 접근이 필요 → List (ArrayList)
List<User> users = new ArrayList<>();
User user = users.get(5);  // O(1) 접근

// 2. 중복 제거가 필요 → Set
Set<String> uniqueWords = new HashSet<>();
uniqueWords.add("hello");  // 중복 자동 무시

// 3. 키-값 매핑이 필요 → Map
Map<String, User> userById = new HashMap<>();
User user = userById.get("user123");  // O(1) 조회

// 4. 정렬된 순서 유지가 필요 → TreeSet/TreeMap
Set<Integer> sortedNumbers = new TreeSet<>();
// 자동으로 정렬된 순서 유지

// 5. 우선순위 기반 처리 → PriorityQueue
PriorityQueue<Task> tasks = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority)
);
Task nextTask = tasks.poll();  // 최소 우선순위 추출

// 6. LIFO 순서 → Deque as Stack
Deque<String> stack = new ArrayDeque<>();
stack.push("item");
String item = stack.pop();

// 7. FIFO 순서 → Queue
Queue<String> queue = new LinkedList<>();
queue.offer("item");
String item = queue.poll();
```

### 인터페이스 기반 프로그래밍

```java
// Good: 인터페이스 타입 사용
public class UserService {
    private final Map<String, User> users;  // 인터페이스 타입

    public UserService(Map<String, User> users) {
        this.users = users;  // 의존성 주입으로 구현 교체 가능
    }
}

// Bad: 구체 클래스 타입 사용
public class UserService {
    private final HashMap<String, User> users;  // 구체 타입에 종속
}
```

## 참고 자료

- Introduction to Algorithms (CLRS), Chapter 10: Elementary Data Structures
- Data Structures and Algorithms in Java, Goodrich et al.
- Java Collections Framework Documentation
- Liskov, Guttag - "Program Development in Java: Abstraction, Specification, and Object-Oriented Design"

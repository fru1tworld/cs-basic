# 동적 배열 (Dynamic Array)

## 개요

동적 배열은 **크기가 자동으로 조절되는 배열**이다. 고정 크기 배열의 한계를 극복하여 원소를 추가할 때 필요에 따라 용량을 확장한다. Java의 ArrayList, Python의 list, C++의 std::vector가 대표적인 구현이다. 삽입 연산은 amortized O(1)의 시간 복잡도를 가진다.

## 핵심 개념

### 동적 배열의 구조

```java
public class DynamicArray<T> {
    private Object[] array;   // 내부 고정 배열
    private int size;         // 실제 원소 개수
    private int capacity;     // 배열의 용량

    public DynamicArray() {
        capacity = 10;  // 초기 용량
        array = new Object[capacity];
        size = 0;
    }

    // size: 실제 저장된 원소 수
    // capacity: 할당된 공간 크기
    // capacity >= size 항상 성립
}
```

```
동적 배열 상태 예시:
size = 5, capacity = 8

┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  1  │  2  │  3  │  4  │  5  │     │     │     │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]
└───────── 사용 중 ─────────┘ └── 예비 공간 ──┘
```

### Growth Factor (성장 계수)

배열이 가득 찼을 때 새 배열의 크기를 결정하는 비율

```java
// Growth Factor = 2 (Java ArrayList, C++ vector)
void grow() {
    int newCapacity = capacity * 2;
    Object[] newArray = new Object[newCapacity];
    System.arraycopy(array, 0, newArray, 0, size);
    array = newArray;
    capacity = newCapacity;
}

// Growth Factor = 1.5 (일부 구현)
void grow() {
    int newCapacity = capacity + capacity / 2;  // 1.5배
    // ...
}
```

**Growth Factor 선택의 trade-off**:

| Factor | 장점 | 단점 |
|--------|------|------|
| 2배 | 복사 빈도 낮음, 상환 비용 낮음 | 최대 50% 메모리 낭비 |
| 1.5배 | 메모리 효율적 (최대 33% 낭비) | 복사 더 자주 발생 |
| 골든 비율 (φ≈1.618) | 이론적 최적, 메모리 재사용 가능 | 실제 효과 미미 |

### Amortized O(1) 삽입 증명

**목표**: n번의 삽입 연산에서 총 비용이 O(n)임을 증명

**증명 (Aggregate Method)**:
```
n번 삽입 시 확장이 발생하는 시점: 1, 2, 4, 8, 16, ..., 2^k (where 2^k ≤ n)

총 복사 비용:
= 1 + 2 + 4 + 8 + ... + 2^k
= 2^(k+1) - 1
< 2n  (since 2^k ≤ n < 2^(k+1))

일반 삽입 비용 = n

총 비용 T(n) < 2n + n = 3n
상환 비용 = T(n)/n < 3 = O(1)
```

**증명 (Accounting Method)**:
```
각 삽입에 3 단위의 비용을 할당:
- 1 단위: 실제 삽입
- 1 단위: 나중에 자신이 복사될 때를 위해 저축
- 1 단위: 이전에 복사된 원소의 복사 비용 보조

확장 시점:
- n/2개의 새 원소가 각각 2단위 저축 = n 단위
- 복사해야 할 원소 = n개
- 저축된 n 단위로 복사 비용 충당 가능

따라서 모든 삽입의 상환 비용 = 3 = O(1)
```

### 삽입 연산 구현

```java
public void add(T element) {
    // 용량 확인 및 확장
    if (size == capacity) {
        grow();  // O(n) 비용, 하지만 드물게 발생
    }
    array[size++] = element;  // O(1) 비용
}

public void add(int index, T element) {
    if (index < 0 || index > size) {
        throw new IndexOutOfBoundsException();
    }

    if (size == capacity) {
        grow();
    }

    // index 이후 원소들을 한 칸씩 뒤로 이동 - O(n)
    for (int i = size; i > index; i--) {
        array[i] = array[i - 1];
    }
    array[index] = element;
    size++;
}
```

### Shrinking Policy (축소 정책)

메모리 낭비를 줄이기 위해 용량을 줄이는 정책

```java
public T remove(int index) {
    T element = (T) array[index];

    // 원소 이동
    for (int i = index; i < size - 1; i++) {
        array[i] = array[i + 1];
    }
    array[--size] = null;  // 메모리 누수 방지

    // 축소 조건: size가 capacity의 1/4 이하일 때 절반으로 축소
    if (size > 0 && size <= capacity / 4) {
        shrink();
    }

    return element;
}

private void shrink() {
    int newCapacity = capacity / 2;
    Object[] newArray = new Object[newCapacity];
    System.arraycopy(array, 0, newArray, 0, size);
    array = newArray;
    capacity = newCapacity;
}
```

**왜 1/4에서 축소하는가?**
```
1/2에서 축소하면 문제 발생:

capacity = 8, size = 4 상태에서:
- add → 확장 → capacity = 16
- remove → 축소 → capacity = 8
- add → 확장 → capacity = 16
- ...
→ thrashing 발생! 매 연산이 O(n)

1/4에서 축소하면:
- 확장 후 size = capacity/2
- size가 capacity/4가 되려면 capacity/4번 삭제 필요
- 그동안 O(1) 삭제가 보장됨
→ amortized O(1) 유지
```

### std::vector / ArrayList 구현 분석

#### Java ArrayList

```java
// Java ArrayList의 핵심 구현 (OpenJDK 기준)
public class ArrayList<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private Object[] elementData;
    private int size;

    // 확장: 약 1.5배 (oldCapacity + oldCapacity >> 1)
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5배

        if (newCapacity < minCapacity)
            newCapacity = minCapacity;

        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    // 삽입 위치 지정 시 arraycopy로 이동
    public void add(int index, E element) {
        ensureCapacityInternal(size + 1);
        System.arraycopy(elementData, index,
                        elementData, index + 1,
                        size - index);
        elementData[index] = element;
        size++;
    }
}
```

#### C++ std::vector

```cpp
// C++ vector의 핵심 동작
template<typename T>
class vector {
private:
    T* data;
    size_t size_;
    size_t capacity_;

public:
    // push_back: amortized O(1)
    void push_back(const T& value) {
        if (size_ == capacity_) {
            // 확장: 보통 2배
            size_t new_capacity = (capacity_ == 0) ? 1 : capacity_ * 2;
            reserve(new_capacity);
        }
        data[size_++] = value;
    }

    // reserve: 미리 용량 확보
    void reserve(size_t new_capacity) {
        if (new_capacity > capacity_) {
            T* new_data = new T[new_capacity];
            std::copy(data, data + size_, new_data);
            delete[] data;
            data = new_data;
            capacity_ = new_capacity;
        }
    }

    // shrink_to_fit: 남는 공간 제거 (C++11)
    void shrink_to_fit() {
        if (size_ < capacity_) {
            T* new_data = new T[size_];
            std::copy(data, data + size_, new_data);
            delete[] data;
            data = new_data;
            capacity_ = size_;
        }
    }
};
```

### 연산별 시간 복잡도

| 연산 | 평균 | 최악 | 비고 |
|------|------|------|------|
| get(i) | O(1) | O(1) | 인덱스 접근 |
| set(i, e) | O(1) | O(1) | 인덱스 접근 |
| add(e) (끝) | O(1) amortized | O(n) | 확장 시 |
| add(i, e) | O(n) | O(n) | 원소 이동 |
| remove(끝) | O(1) | O(1) | |
| remove(i) | O(n) | O(n) | 원소 이동 |
| contains(e) | O(n) | O(n) | 선형 탐색 |

## 실무 적용

### 용량 사전 할당

```java
// Bad: 기본 용량으로 시작 → 확장 여러 번
List<String> list = new ArrayList<>();
for (int i = 0; i < 10000; i++) {
    list.add("item" + i);  // 여러 번 확장 발생
}

// Good: 예상 크기로 초기화
List<String> list = new ArrayList<>(10000);
for (int i = 0; i < 10000; i++) {
    list.add("item" + i);  // 확장 없음
}

// 확장 횟수 비교 (기본 용량 10, growth factor 1.5):
// Bad: 10 → 15 → 22 → 33 → ... → 10000 (약 24번 확장)
// Good: 확장 없음
```

### trimToSize()

```java
// 대량 데이터 처리 후 메모리 절약
List<Integer> list = new ArrayList<>(100000);
// ... 데이터 처리 ...
// 최종적으로 1000개만 남음

// 99000개의 빈 슬롯이 메모리 낭비
// trimToSize()로 용량을 size에 맞춤
((ArrayList<Integer>) list).trimToSize();
```

### 배열 vs ArrayList 선택

```java
// 배열이 적합한 경우:
// 1. 크기가 고정된 경우
// 2. 기본 타입 대량 저장 (메모리/성능)
// 3. 다차원 데이터

int[] scores = new int[100];  // 고정 크기
int[][] matrix = new int[1000][1000];  // 2D 데이터

// ArrayList가 적합한 경우:
// 1. 크기가 변하는 경우
// 2. 편의 메서드가 필요한 경우 (contains, indexOf 등)
// 3. 제네릭 타입 안전성이 필요한 경우

List<User> users = new ArrayList<>();
users.add(new User("Alice"));
users.removeIf(u -> u.isInactive());
```

## 참고 자료

- Introduction to Algorithms (CLRS), Chapter 17: Amortized Analysis
- Java ArrayList Source Code (OpenJDK)
- C++ STL vector Implementation
- The Art of Computer Programming Vol. 1 - Knuth

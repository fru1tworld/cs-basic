# 분할 상환 분석 (Amortized Analysis)

## 개요

분할 상환 분석은 일련의 연산에서 **평균적인 연산 비용**을 분석하는 기법이다. 개별 연산은 비용이 클 수 있지만, 전체 연산 시퀀스에서 평균을 내면 각 연산의 비용이 낮아지는 것을 증명한다. 최악의 경우 분석보다 더 정확한 성능 예측이 가능하다.

### 평균 케이스 분석과의 차이

- **평균 케이스 분석**: 확률적 가정에 기반한 기대값
- **분할 상환 분석**: 최악의 경우에도 보장되는 평균 (확률 가정 없음)

## 핵심 개념

### 세 가지 분석 방법

```
1. Aggregate Method (총합 분석)
2. Accounting Method (회계 분석)
3. Potential Method (포텐셜 분석)
```

---

### 1. Aggregate Method (총합 분석)

**아이디어**: n개 연산의 총 비용 T(n)을 계산하고, 평균 비용 T(n)/n을 구함

#### 예시: 동적 배열의 삽입

```java
class DynamicArray {
    private int[] array;
    private int size;
    private int capacity;

    public DynamicArray() {
        capacity = 1;
        array = new int[capacity];
        size = 0;
    }

    // 배열이 가득 차면 2배로 확장
    public void add(int element) {
        if (size == capacity) {
            // 확장: O(n) 비용
            capacity *= 2;
            int[] newArray = new int[capacity];
            for (int i = 0; i < size; i++) {
                newArray[i] = array[i];
            }
            array = newArray;
        }
        array[size++] = element;  // O(1) 비용
    }
}
```

**분석**:
```
n번의 삽입 연산 비용:
- 확장이 발생하는 시점: 1, 2, 4, 8, ..., 2^k (2^k ≤ n < 2^(k+1))
- 확장 비용: 1 + 2 + 4 + 8 + ... + 2^k = 2^(k+1) - 1 < 2n
- 일반 삽입 비용: n

총 비용 T(n) < 2n + n = 3n
분할 상환 비용 = T(n)/n < 3 = O(1)
```

#### 예시: 이진 카운터 증가

```java
class BinaryCounter {
    private int[] bits;  // bits[0]이 LSB

    public BinaryCounter(int k) {
        bits = new int[k];  // k비트 카운터
    }

    // 1 증가
    public void increment() {
        int i = 0;
        while (i < bits.length && bits[i] == 1) {
            bits[i] = 0;  // flip
            i++;
        }
        if (i < bits.length) {
            bits[i] = 1;  // flip
        }
    }
}
```

**분석**:
```
n번의 increment 연산에서 각 비트가 flip되는 횟수:
- bits[0]: 매번 flip → n번
- bits[1]: 2번에 1번 flip → n/2번
- bits[2]: 4번에 1번 flip → n/4번
- ...
- bits[k-1]: 최대 n/2^(k-1)번

총 flip 횟수 = n + n/2 + n/4 + ... < 2n
분할 상환 비용 = T(n)/n < 2 = O(1)
```

---

### 2. Accounting Method (회계 분석)

**아이디어**: 각 연산에 "상환 비용(amortized cost)"을 할당하고, 실제 비용보다 많이 지불한 것은 "크레딧"으로 저축하여 나중에 비싼 연산에 사용

**규칙**:
- 상환 비용 ≥ 실제 비용의 합 (항상)
- 크레딧은 음수가 되면 안 됨

#### 예시: 동적 배열의 삽입

```
연산별 비용 설정:
- 삽입 1회당 상환 비용: 3 단위

비용 분배:
- 실제 삽입: 1 단위 (즉시 사용)
- 나중 복사를 위한 크레딧: 2 단위 (저축)
  - 자신을 복사할 비용: 1 단위
  - 이전 원소의 복사 도움: 1 단위

검증:
- 확장 시점에 원소 수가 k → 2k로 증가
- k개 원소 복사 비용 = k
- 저축된 크레딧 = 이전 확장 이후 삽입된 k/2개 × 2 = k
- 크레딧으로 정확히 복사 비용 지불 가능
```

```java
/**
 * 회계 분석 관점의 동적 배열
 *
 * 상환 비용 = 3 (실제 삽입 1 + 크레딧 2)
 *
 * 크레딧 사용:
 *   [1][2][3][4]   capacity=4, 4번째까지 삽입 완료
 *                  크레딧: 원소 3,4가 각각 2씩 = 4
 *
 *   [1][2][3][4][5][6][7][8]   확장 후 5~8 삽입
 *   └─복사비용: 4 (크레딧 4로 지불)
 */
```

#### 예시: 스택의 MultiPop

```java
class AmortizedStack {
    private Stack<Integer> stack = new Stack<>();

    // O(1) 실제 비용
    public void push(int x) {
        stack.push(x);
    }

    // O(1) 실제 비용
    public int pop() {
        return stack.pop();
    }

    // O(k) 실제 비용 (k = min(k, size))
    public void multiPop(int k) {
        while (!stack.isEmpty() && k > 0) {
            stack.pop();
            k--;
        }
    }
}
```

**회계 분석**:
```
상환 비용 할당:
- push: 2 (실제 1 + 나중 pop을 위한 크레딧 1)
- pop: 0 (크레딧 사용)
- multiPop(k): 0 (크레딧 사용)

검증:
- 각 원소는 push될 때 2를 지불
- pop/multiPop에서 pop당 1의 크레딧 사용
- 원소는 최대 1번만 pop되므로 크레딧 충분

n개 연산의 총 상환 비용 ≤ 2n (push가 최대 n번)
분할 상환 비용 = O(1)
```

---

### 3. Potential Method (포텐셜 분석)

**아이디어**: 자료구조의 "포텐셜(잠재 에너지)"을 정의하고, 상환 비용 = 실제 비용 + 포텐셜 변화

```
ĉᵢ = cᵢ + Φ(Dᵢ) - Φ(Dᵢ₋₁)

여기서:
- ĉᵢ: i번째 연산의 상환 비용
- cᵢ: i번째 연산의 실제 비용
- Φ(Dᵢ): i번째 연산 후 자료구조의 포텐셜
- Φ(D₀) = 0 (초기 포텐셜)
- Φ(Dᵢ) ≥ 0 (모든 i에 대해)
```

**총 상환 비용**:
```
Σĉᵢ = Σcᵢ + Φ(Dₙ) - Φ(D₀)
    = Σcᵢ + Φ(Dₙ)  (Φ(D₀) = 0이므로)
    ≥ Σcᵢ          (Φ(Dₙ) ≥ 0이므로)
```

#### 예시: 동적 배열

```
포텐셜 함수 정의:
Φ(D) = 2 × size - capacity

검증:
- 확장 직전: size = capacity = n
  Φ = 2n - n = n

- 확장 직후: size = n, capacity = 2n
  Φ = 2n - 2n = 0

일반 삽입의 상환 비용:
ĉ = 1 + (2(size+1) - capacity) - (2×size - capacity)
  = 1 + 2 = 3

확장 삽입의 상환 비용:
ĉ = (1 + n) + (2(n+1) - 2n) - (2n - n)
  = n + 1 + 2 - n = 3

모든 삽입의 상환 비용 = O(1)
```

#### 예시: Splay Tree의 Splay 연산

```
포텐셜 함수 정의:
Φ(T) = Σ rank(x)  (모든 노드 x에 대해)

여기서 rank(x) = log(size(x))
size(x) = x를 루트로 하는 서브트리의 노드 수

Access Lemma:
splay(x)의 상환 비용 ≤ 3(rank(root) - rank(x)) + 1
                    = O(log n)

증명은 Zig, Zig-Zig, Zig-Zag 각 케이스 분석으로 진행
(상세 증명은 CLRS 17장 참조)
```

---

### 분석 방법 비교

| 방법 | 장점 | 단점 | 적합한 경우 |
|------|------|------|-------------|
| Aggregate | 직관적, 단순 | 개별 연산 비용 구분 어려움 | 모든 연산이 동일한 경우 |
| Accounting | 개별 연산 분석 가능 | 적절한 크레딧 설정 필요 | 직관적으로 "저축" 설명 가능한 경우 |
| Potential | 가장 일반적, 강력 | 포텐셜 함수 설계 어려움 | 복잡한 자료구조 분석 |

## 실무 적용

### 동적 배열의 Growth Factor 선택

```java
// Growth Factor = 2 (Java ArrayList, C++ vector)
// 장점: 분할 상환 비용이 낮음
// 단점: 메모리 낭비 최대 50%

// Growth Factor = 1.5 (일부 구현)
// 장점: 메모리 효율적
// 단점: 복사가 더 자주 발생

/**
 * Growth Factor에 따른 분할 상환 분석
 *
 * Factor = k (k > 1)일 때:
 * - n번 삽입 시 확장 횟수 ≈ log_k(n)
 * - 총 복사 비용 = n + n/k + n/k² + ... = n × k/(k-1)
 * - 상환 비용 = k/(k-1)
 *
 * k=2: 상환 비용 = 2
 * k=1.5: 상환 비용 = 3
 */
```

### HashMap의 Rehashing

```java
/**
 * HashMap은 load factor 초과 시 rehash
 * 기본 load factor = 0.75
 *
 * Rehash 비용:
 * - 새 배열 할당: O(n)
 * - 모든 엔트리 재배치: O(n)
 *
 * 분할 상환 분석:
 * - n개 삽입 시 rehash 횟수 ≈ log(n)
 * - 총 rehash 비용 < 2n
 * - 삽입 상환 비용 = O(1)
 */
public class HashMapAnalysis {
    // 실제로 Java HashMap은 treeify(O(log n))도 있지만
    // 여전히 삽입은 amortized O(1)
}
```

### StringBuilder vs String 연결

```java
// String 연결: O(n²) - 매번 새 String 생성
String result = "";
for (int i = 0; i < n; i++) {
    result += "x";  // O(i) 복사 발생
}
// 총 비용: 1 + 2 + 3 + ... + n = O(n²)

// StringBuilder: O(n) - 분할 상환 O(1) 삽입
StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) {
    sb.append("x");  // amortized O(1)
}
String result = sb.toString();
// 총 비용: O(n)
```

## 참고 자료

- Introduction to Algorithms (CLRS), Chapter 17: Amortized Analysis
- MIT 6.046J Lecture on Amortized Analysis
- Sleator, Tarjan - "Self-Adjusting Binary Search Trees" (Splay Tree 포텐셜 분석)
